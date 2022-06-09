---
title: "Ceph 疑难杂症"
description : "Ceph 遇到到一些问题"
isCJKLanguage: true
date: 2022-06-09T11:25:23+08:00
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**

# OSD 使用太多的内存

## 参考

[OSD and MON memory consumption](https://github.com/rook/rook/issues/5811)  
[Ceph OSD Pod memory consumption very high](https://github.com/rook/rook/issues/5821)  
[Ceph Cluster CRD](https://rook.github.io/docs/rook/v1.7/ceph-cluster-crd.html)  
[Ceph HardWare Recommendations](https://docs.ceph.com/en/latest/start/hardware-recommendations/)

## 实操

### 现在的情况

> 可以看到 ceph-osd 占用了大量到内存
```shell
# top -p $(pidof ceph-osd)
top - 11:30:08 up 26 days, 16:38,  3 users,  load average: 0.74, 0.57, 0.72
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.5 us,  1.6 sy,  0.0 ni, 94.1 id,  1.3 wa,  0.0 hi,  0.5 si,  0.0 st
MiB Mem : 91.5/16008.3  [||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||        ]
MiB Swap:  0.0/0.0      [                                                                                                    ]

    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND                                                                                        
   5831 167       20   0   12.7g  11.0g   7720 S   2.7  70.5 617:42.90 ceph-osd             


# kubectl -n rook-ceph  exec -it rook-ceph-tools-6ccb958485-j7pvb  -- bash
# ceph osd tree
ID  CLASS  WEIGHT   TYPE NAME       STATUS  REWEIGHT  PRI-AFF
-1         3.00000  root default                             
-3         1.00000      host node1                           
 1    hdd  1.00000          osd.1       up   1.00000  1.00000
-7         1.00000      host node2                           
 2    hdd  1.00000          osd.2       up   1.00000  1.00000
-5         1.00000      host node3                           
 0    hdd  1.00000          osd.0       up   1.00000  1.00000
### 太残忍了， 就 15，6G 的内存你就要 13G， 还要不要其他进程活了？
# ceph tell osd.0 config show | grep -w osd_memory_target
    "osd_memory_target": "13344836812",
# ceph tell osd.1 config show | grep -w osd_memory_target
    "osd_memory_target": "13344836812",
# ceph tell osd.2 config show | grep -w osd_memory_target
    "osd_memory_target": "13344836812",
### 安？ 没有限制资源？难怪敢用那么多的内存。
# kubectl  -n rook-ceph get pods rook-ceph-osd-0-7c76474f7-tnhc6 -ojson | jq .spec.containers[].resources
{}
```

### 资源限制
> 按照建议限制个 2cpu 和 4G 内存
```shell
## 编辑 CR， 添加资源限制（会自动重新配置 osd）
# kubectl -n rook-ceph  edit  cephclusters.ceph.rook.io rook-ceph
```
```yaml
  storage:
    nodes:
    - devices:
      - name: sdb
      name: node1
      resources:
        limits:
          cpu: "2"
          memory: "4096Mi"
        requests:
          cpu: "2"
          memory: "4096Mi"
    - devices:
      - name: sdb
      name: node2
      resources:
        limits:
          cpu: "2"
          memory: "4096Mi"
        requests:
          cpu: "2"
          memory: "4096Mi"
    - devices:
      - name: sdb
      name: node3
      resources:
        limits:
          cpu: "2"
          memory: "4096Mi"
        requests:
          cpu: "2"
          memory: "4096Mi"
```

### 再次看看现在到情况
> 已经有限制了
```shell
# kubectl  -n rook-ceph get pods rook-ceph-osd-0-6ff54bb9c7-vbk59 -ojson | jq .spec.containers[].resources
```
```json
{
  "limits": {
    "cpu": "2",
    "memory": "4Gi"
  },
  "requests": {
    "cpu": "2",
    "memory": "4Gi"
  }
}
```

```shell
### 这下看着舒服了
# top -p $(pidof ceph-osd)
top - 11:50:13 up 26 days, 16:58,  3 users,  load average: 0.71, 0.97, 0.89
Tasks:   1 total,   0 running,   1 sleeping,   0 stopped,   0 zombie
%Cpu(s):  2.7 us,  1.1 sy,  0.0 ni, 96.0 id,  0.1 wa,  0.0 hi,  0.1 si,  0.0 st
MiB Mem : 23.6/16008.3  [||||||||||||||||||||||||]
MiB Swap:  0.0/0.[]
    PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND 
2172679 167       20   0 1462020 455712  33268 S   2.3   2.8   0:11.17 ceph-osd 

# kubectl -n rook-ceph  exec -it rook-ceph-tools-6ccb958485-j7pvb  -- bash
# ceph tell osd.0 config show | grep -w osd_memory_target
    "osd_memory_target": "3435973836",
```