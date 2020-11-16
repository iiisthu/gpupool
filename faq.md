# FAQ

这里列举一些常见问题供大家参考。

## Apply后显示deployment已创建，但是没有Pods

POD 超过默认的资源配额或者集群资源不足。首先参考[资源配额](#资源配额)修改配额配置（可以通过`kubectl describe resourcequota xxx`查看具体的配置）。

## 删除 Pod 后，`kubectl get pods`又出现新的Pod

应该删除`deployment`资源。我们通过申请`deployment`资源会创建 POD，同时监控是否有 POD 挂掉，如果挂掉了会再向集群申请资源，始终维持可用的 POD 个数保持在配置文件中`replicas`指定的备份数目。

```bash
kubectl delete deployment my-first-ubuntu-tf
```

## 创建 POD 失败，提示资源不足

> **_NOTE:_** 默认不用修改资源配额，如果由于已经存在的资源配额配置导致 POD 启动失败，可尝试以下方法。

当需要更改计算资源时可以应用新的资源配额配置（参考`quota.yaml`）。首先删除旧的资源配额配置：

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
