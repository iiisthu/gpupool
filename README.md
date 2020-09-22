# GPU集群使用

- [环境配置](#环境配置)
    - [配置VPN连接](#配置VPN连接)
    - [kubectl安装](#kubectl安装)
    - [kubeconfig获取](#kubeconfig获取)
- [集群使用](#集群使用)
    - [镜像列表](#镜像列表)
    - [资源配额](#资源配额)
    - [操作容器](#操作容器)
    - [数据、程序和结果的持久化存储](#数据程序和结果的持久化存储)
- [FAQ](#FAQ)

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

|镜像|可用Tag|GPU支持|备注|
|---|----|------|---|
|ubuntu-tensorflow|1.14.1, 2.3.0|是|Tensorflow版本1.14.1/2.3.0，CUDA版本10.1|
|ubuntu-pytorch|1.5.0|是|Pytorch版本1.5.0，CUDA版本10.1|
|orion-client-2.4.2|cu10.0_cudnn7_ubuntu18.04-base, cu10.1_cudnn7_ubuntu18.04-base, cu10.2_cudnn7_ubuntu18.04-base|是|仅包含虚拟GPU的镜像，需要自行安装Tensorflow等库，Tag区别仅是CUDA版本|
|cuda|10.0-cudnn7-devel-ubuntu18.04, 10.1-cudnn7-devel-ubuntu18.04, 10.2-cudnn7-devel-ubuntu18.04|是|仅包含CUDA支持，只能访问物理机的GPU，Tag区别仅是CUDA版本|

在配置文件中通过`172.16.112.173:30006/library/<镜像>:<Tag>`从指定镜像版本创建 POD。为了方便，这里列出yaml中镜像一行的常用选择:

```yaml
image: 172.16.112.173:30006/library/ubuntu-tensorflow:2.3.0
image: 172.16.112.173:30006/library/ubuntu-tensorflow:1.14.1
image: 172.16.112.173:30006/library/ubuntu-pytorch:1.5.0
```

### 资源配额

> **_NOTE:_** 未来默认不用修改资源配额，如果由于已经存在的资源配额配置导致 POD 启动失败，可尝试以下方法。

默认每个用户的命名空间下申请的每个容器的最大资源占用为10个GPU，10G的内存。当需要更大的计算资源时可以应用新的资源配额配置（参考`quota.yaml`）。首先删除旧的资源配额配置：

```bash
# 查看当前所有的资源配额配置
$ kubectl get resourcequotas
NAME                          CREATED AT
compute-resources-zhangsan1   2020-08-27T08:44:25Z

# 删除已有资源配额配置
$ kubectl delete resourcequotas compute-resources-zhangsan1
resourcequota "compute-resources-zhangsan1" deleted

# 应用新的配置
$ kubectl apply -f quota.yaml
resourcequota/mem-cpu-quota created
```

目前给出的所有的yaml模板均使用`Mem 20G + 10 CPU`的配置，因此如果 POD 不能成功创建，需要先应用新的配额配置。

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

### 数据、程序和结果的持久化存储

POD 的本地文件是临时的，在每次重启（手动或失败重启）后都会恢复到最初的镜像状态。为了保持在 POD 中做的状态变化（例如创建了数据文件、日志文件等），需要向集群申请持久化存储资源。目前提供两种持久化存储方案：直接挂载NFS的共享盘和申请[PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，前者用于跨用户共享，后者为个人可见。两者均为网络存储，因此效率有限。我们在每台物理机上配置了两块3T的SSD硬盘作为本地缓存用，建议两者配合使用（见[挂载本地缓存SSD](#挂载本地缓存SSD)）。

> **_NOTE:_** 提供持久化存储服务的硬盘并不保证和 POD 处于同一台物理机上，因此如果有大量的文件IO，建议先将数据拷贝到 POD 同一台物理机的磁盘，再从本地磁盘读写。

#### 直接挂载 NFS 共享盘

集群 NFS 向所有用户提供了持久化的共享存储空间，可以理解为一个远程的目录。这个目录的主要目的是共享一些公共的数据集（例如某些benchmark训练数据之类的），方便大家共同使用。这一存储的优点是可靠性高（后台是华为的专用存储阵列，带RAID），缺点是性能比较低，因此要尽量避免直接在上边进行大规模的数据读写操作。

把数据集放到共享目录下的方法：请每位用户自行以自己分配到的集群命名空间为名建立文件夹（例如命名空间为`zhangsan1`则建立`share/zhangsan1`），公开数据集请放入`share/datasets/`下，并附带说明文件README指出数据源及具体描述。挂载NFS共享盘的方式：

```bash
kubectl apply -f ubuntu-tf-nfs-direct-example.yaml
```

> **_NOTE:_** 使用时尽量把数据拷贝到下边的本地磁盘再使用！

如果西安集群访问Internet非常慢，数据集下载不下来等情况，请联系尹伟老师，他会通过其他途径把数据集放到这个共享盘上。

> **_WARNING:_** 严禁任意修改、删除他人的工作目录！

#### 申请NFS上的独立存储盘PVC

如果你需要一个自己的共享文件目录，不希望被别人看见、或者能被别人删除的，例如可以存自己的代码啊，程序运行的结果等内容。这个盘的后端跟上述共享目录是同一套存储，也具有比较好的可靠性，但是同样的，不要指望高性能，因此也不要在上边分析大量的数据。

使用 PVC 和 POD 相似，都向集群申请临时资源，不同的是 PVC 申请的是存储资源，POD 申请的是计算资源。PVC 和 POD 的生命周期是独立的，重启 POD 后 PVC 中的数据并不会消失。通过 PVC 获得持久化存储：

```bash
kubectl apply -f nfs-pvc-example.yaml
```
在 POD 中挂载 PVC：

```bash
kubectl apply -f ubuntu-tf+pvc-example.yaml
```

#### 挂载本地缓存SSD

> **_NOTE:_** 本地磁盘只做临时缓存用，并不保证容错，每次重启POD的时候，可能会被清空。

每台物理机本地有两块SSD可供挂载，读写速度会比 NFS 共享盘和 PVC 高很多。实测性能请见下边的测试，两块盘的每秒顺序读写分别超过了2GB和3GB。
建议和持久盘配合使用，推荐的用法是：

* 启动POD的时候，把需要的数据从上边的share磁盘或者自己的PVC拷贝到这个缓存 （可以用rsync命令）；
* 用缓存的数据来跑程序，结果也写到这个缓存中；
* 关闭POD前，把结果拷贝到PVC中去（用rsync命令）。

挂载本地磁盘参考

```bash
kubectl apply -f ubuntu-tf+local-disk-example.yaml
```

在这个模板中，两个SSD盘分别挂在`/mnt/data1`和`/mnt/data2`两个目录下。

> **_NOTE:_** 想要同时挂载多个卷（如同时挂载SSD和NFS共享盘），请参考各个例子中声明volume的部分，组合起来即可。

#### 实测POD内存储性能

用[fio](https://github.com/axboe/fio/tree/master/examples)测试了本地SSD盘的速度

|目录|参数|性能|
|---|---|---|
|`/share`|写入|2457600000 bytes (2.5 GB, 2.3 GiB) copied, 56.7821 s, 43.3 MB/s|
|`/share`|读取|2457600000 bytes (2.5 GB, 2.3 GiB) copied, 33.293 s, 73.8 MB/s|
|`/mnt/data1`|fio-seq-write.fio|bw=2825MiB/s (2963MB/s), 2825MiB/s-2825MiB/s (2963MB/s-2963MB/s)|
|`/mnt/data1`|fio-seq-read.fio|bw=2147MiB/s (2252MB/s), 2147MiB/s-2147MiB/s (2252MB/s-2252MB/s)|
|`/mnt/data2`|fio-seq-write.fio|bw=3130MiB/s (3282MB/s), 3130MiB/s-3130MiB/s (3282MB/s-3282MB/s)|
|`/mnt/data2`|fio-seq-read.fio|bw=3082MiB/s (3232MB/s), 3082MiB/s-3082MiB/s (3232MB/s-3232MB/s)|

## FAQ

这里列举一些常见问题供大家参考。

### Apply后显示deployment已创建，但是没有Pods

POD 超过默认的资源配额或者集群资源不足。首先参考[资源配额](#资源配额)修改配额配置（可以通过`kubectl describe resourcequota xxx`查看具体的配置）。

### 删除 Pod 后，`kubectl get pods`又出现新的Pod

应该删除`deployment`资源。我们通过申请`deployment`资源会创建 POD，同时监控是否有 POD 挂掉，如果挂掉了会再向集群申请资源，始终维持可用的 POD 个数保持在配置文件中`replicas`指定的备份数目。

```bash
kubectl delete deployment my-first-ubuntu-tf
```
