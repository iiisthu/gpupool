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

## 基于 orion-client-2.4.2 打的镜像，如何安装 Pytorch 或者 Tensorflow

具体用`pip`安装的指令见[这里](https://pypi.virtaitech.com/)。

> **_NOTE:_** CUDA 版本选择10.1。

## 可以连接VPN，但是无法访问 harbor.iiis.co

部分路由器可能会禁止指向本地IP的域名，若能连接 VPN，可以尝试直接访问IP的方式，跳过域名查询。首先查询harbor.iiis.co对应的IP：

```bash
$ dig harbor.iiis.co
; <<>> DiG 9.10.6 <<>> harbor.iiis.co

;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 40516
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;harbor.iiis.co.			IN	A

;; ANSWER SECTION:
harbor.iiis.co.		1800	IN	A	172.16.112.220

;; Query time: 331 msec
;; SERVER: 221.179.155.177#53(221.179.155.177)
;; WHEN: Fri Dec 18 22:54:11 CST 2020
;; MSG SIZE  rcvd: 48
```

因此我们知道IP为 172.16.112.220。因此可以尝试访问 172.16.112.220:31388。
