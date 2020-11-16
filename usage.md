# 集群使用

这里我们介绍 GPU 集群的基本使用，在使用之前需要正确[配置集群环境](https://github.com/iiisthu/gpupool/blob/master/environment.md)。

> **_NOTE:_** 集群的连接均采用命令行工具`kubectl`，当使用遇见问题时建议阅读详细的 `kubectl` 教程 [kubectl book](https://kubectl.docs.kubernetes.io/)。

当创建了集群账号后集群会自动给用户分配[命名空间](http://harbor.iiis.co:31388/)，在连接集群时需要指定具体的 namespace，否则会提示权限错误。为了避免每次均输入命名空间参数，可以利用`alias`：

```bash
# .bashrc
alias kubectl="kubectl -n zhangsan"  # zhangsan为分配的namespace
```

之后都默认在自己的 namespace 下操作。

## 镜像列表

Docker 镜像仓库位置：`harbor.iiis.co/library/`。以下是提供的支持GPU的镜像列表，其他镜像可以从 Docker Hub 拉取。

|镜像|可用Tag|GPU支持|备注|
|---|----|------|---|
|ubuntu-tensorflow|1.14.1, 2.3.0|是|Tensorflow版本1.14.1/2.3.0，CUDA版本10.1|
|ubuntu-pytorch|1.5.0|是|Pytorch版本1.5.0，CUDA版本10.1|
|orion-client-2.4.2|cu10.0_cudnn7_ubuntu18.04-base, cu10.1_cudnn7_ubuntu18.04-base, cu10.2_cudnn7_ubuntu18.04-base|是|仅包含虚拟GPU的镜像，需要自行安装Tensorflow等库，Tag区别仅是CUDA版本|
|cuda|10.0-cudnn7-devel-ubuntu18.04, 10.1-cudnn7-devel-ubuntu18.04, 10.2-cudnn7-devel-ubuntu18.04|是|仅包含CUDA支持，只能访问物理机的GPU，Tag区别仅是CUDA版本|

在配置文件中通过`harbor.iiis.co/library/<镜像>:<Tag>`从指定镜像版本创建 POD。为了方便，这里列出yaml中镜像一行的常用选择:

```yaml
image: harbor.iiis.co/library/ubuntu-tensorflow:2.3.0
image: harbor.iiis.co/library/ubuntu-tensorflow:1.14.1
image: harbor.iiis.co/library/ubuntu-pytorch:1.5.0
```

## 操作容器

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

## 数据、程序和结果的持久化存储

POD 的本地文件是临时的，在每次重启（手动或失败重启）后都会恢复到最初的镜像状态。为了保持在 POD 中做的状态变化（例如创建了数据文件、日志文件等），需要向集群申请持久化存储资源。目前提供两种持久化存储方案：直接挂载NFS的共享盘和申请[PVC](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)，前者用于跨用户共享，后者为个人可见。两者均为网络存储，因此效率有限。我们在每台物理机上配置了两块3T的SSD硬盘作为本地缓存用，建议两者配合使用（见[挂载本地缓存SSD](#挂载本地缓存SSD)）。

> **_NOTE:_** 提供持久化存储服务的硬盘并不保证和 POD 处于同一台物理机上，因此如果有大量的文件IO，建议先将数据拷贝到 POD 同一台物理机的磁盘，再从本地磁盘读写。

### 直接挂载 NFS 共享盘

集群 NFS 向所有用户提供了持久化的共享存储空间，可以理解为一个远程的目录。这个目录的主要目的是共享一些公共的数据集（例如某些benchmark训练数据之类的），方便大家共同使用。这一存储的优点是可靠性高（后台是华为的专用存储阵列，带RAID），缺点是性能比较低，因此要尽量避免直接在上边进行大规模的数据读写操作。

把数据集放到共享目录下的方法：请每位用户自行以自己分配到的集群命名空间为名建立文件夹（例如命名空间为`zhangsan1`则建立`share/zhangsan1`），公开数据集请放入`sharenfs/datasets/`下，并附带说明文件README指出数据源及具体描述。挂载NFS共享盘的方式：

```bash
kubectl apply -f ubuntu-tf-nfs-direct-example.yaml
```

> **_NOTE:_** 使用时尽量把数据拷贝到下边的本地磁盘再使用！

如果西安集群访问Internet非常慢，数据集下载不下来等情况，请联系尹伟老师，他会通过其他途径把数据集放到这个共享盘上。

> **_WARNING:_** 严禁任意修改、删除他人的工作目录！

### 申请NFS上的独立存储盘PVC

如果你需要一个自己的共享文件目录，不希望被别人看见、或者能被别人删除的，例如可以存自己的代码啊，程序运行的结果等内容。这个盘的后端跟上述共享目录是同一套存储，也具有比较好的可靠性，但是同样的，不要指望高性能，因此也不要在上边分析大量的数据。

使用 PVC 和 POD 相似，都向集群申请临时资源，不同的是 PVC 申请的是存储资源，POD 申请的是计算资源。PVC 和 POD 的生命周期是独立的，重启 POD 后 PVC 中的数据并不会消失。通过 PVC 获得持久化存储：

```bash
kubectl apply -f nfs-pvc-example.yaml
```
在 POD 中挂载 PVC：

```bash
kubectl apply -f ubuntu-tf+pvc-example.yaml
```

### 挂载本地缓存SSD

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