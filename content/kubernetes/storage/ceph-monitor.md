---
title: "Move mon to other node in k8s"
description : "迁移 Montir 到其他 Node"
isCJKLanguage: true
date: 2022-05-30T16:20:23+08:00
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**

# 参考
[ Move mon to other node in k8s ](https://github.com/rook/rook/issues/6835)  

# 实操

## 现在的情况

> 可以看到 mon-b 在 node1, mon-c 在 node4, mon-e 在 node2
```shell
$ kubectl -n rook-ceph  get pods -l app=rook-ceph-mon -owide
NAME                               READY   STATUS    RESTARTS   AGE    IP               NODE    NOMINATED NODE   READINESS GATES
rook-ceph-mon-b-84497c47c-pd7lz    1/1     Running   0          76m    10.233.102.142   node1   <none>           <none>
rook-ceph-mon-c-7cb4c5b74f-gv8fb   1/1     Running   0          89m    10.233.74.89     node4   <none>           <none>
rook-ceph-mon-e-66cbb69dcc-nldgq   1/1     Running   0          122m   10.233.75.63     node2   <none>           <none>
```

## 将 mon-c 移动到 node3

> The mon ip address cannot change. If you keep the mon service with the same clusterIP, you could stop the mon and move the /var/lib/rook to a new node if you change the metadata in the rook-ceph-mon-endpoints configmap. You have already added the mon placement to the CephCluster CR? If so, you could also just take one mon down and the operator will start a new mon and failover the old one after 10 minutes on a node that matches the new placement. 可以看出方法有 2， 1 编辑cr, 配置 mon 的 placement， 2 停止 mon， 移动 /var/lib/rook 到新到节点，编辑 configmap rook-ceph-mon-endpoints.

### 更新 cr 文件
> 失败

```shell
kubectl -n rook-ceph  edit  cephclusters.ceph.rook.io rook-ceph
```
```yaml
  placement:
    all:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
    # mon 这一段是添加到
    mon:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: node-role.kubernetes.io/master
                operator: Exists

```
```shell
# 再来验证一下
kubectl -n rook-ceph  edit  cephclusters.ceph.rook.io rook-ceph
```
```yaml
  placement:
    all:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
    # 添加到没有了，换方式 2 吧
    mon: {}
```

### 移动  /var/lib/rook

> 像个永动机一样，应该怎么样把 pod 给停止了呢？

- 停止 mon

```shell
kubectl taint node node4 killer:NoSchedule
node/node4 tainted
kubectl -n rook-ceph delete deployments.apps rook-ceph-mon-c 
deployment.apps "rook-ceph-mon-c" deleted
# 汗颜，这也勉强算停止了 mon 吧
kubectl -n rook-ceph get pods -owide -w | grep mon-c
rook-ceph-mon-c-7cb4c5b74f-ntmc4                  0/1     Pending     0          12s     <none>            <none>   <none>           <none>
```

- 移动 /var/lib/rook

```shell
# 这一步骤是在 node3 上操作
root@node3:/var/lib# scp -r node4:/var/lib/rook /var/lib/
kubectl -n rook-ceph edit configmaps rook-ceph-mon-endpoints
```
```yaml
data:
  csi-cluster-config-json: '[{"clusterID":"rook-ceph","monitors":["10.233.21.50:6789","10.233.34.93:6789","10.233.63.98:6789"]}]'
  data: b=10.233.34.93:6789,c=10.233.63.98:6789,e=10.233.21.50:6789
  # 需要把这里到 node4 相关到改成 node3 相关
  mapping: '{"node":{"b":{"Name":"node1","Hostname":"node1","Address":"192.168.101.101"},"c":{"Name":"node4","Hostname":"node4","Address":"192.168.101.104"},"e":{"Name":"node2","Hostname":"node2","Address":"192.168.101.102"}}}'
  .......
  mapping: '{"node":{"b":{"Name":"node1","Hostname":"node1","Address":"192.168.101.101"},"c":{"Name":"node3","Hostname":"node3","Address":"192.168.101.103"},"e":{"Name":"node2","Hostname":"node2","Address":"192.168.101.102"}}}'

```
```shell
kubectl -n rook-ceph delete deployments.apps rook-ceph-mon-c 
# 已达到预期结果
kubectl -n rook-ceph  get pods -owide | grep mon
rook-ceph-mon-b-84497c47c-pd7lz                   1/1     Running     0          141m    10.233.102.142    node1    <none>           <none>
rook-ceph-mon-c-585ffd65fb-d5dz5                  1/1     Running     0          38s     10.233.71.10      node3    <none>           <none>
rook-ceph-mon-e-66cbb69dcc-nldgq                  1/1     Running     0          3h7m    10.233.75.63      node2    <none>           <none>
# 别忘了把粑粑擦干净
kubectl taint node node4 killer-
node/node4 untainted
ssh root@node4 rm -rf /var/lib/rook
```
