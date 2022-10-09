---
title: "部署 Containerd"
isCJKLanguage: true
date: 2022-10-09T14:10:16+08:00
#lastmod: 2022-04-28T15:13:38+08:00
#categories: [CSI]
#tags: [kubernetes, csi, storage, volume]
draft: false
enableDisqus : true
enableMathJax: false
toc: true
disableToC: false
disableAutoCollapse: true
---

> **Have a nice day! :heart:**
---

# 前言
 从 [kubernetes, docker 渊源](/kubernetes/containerd/difference-docker-containerd-runc-crio-oci/)中可以看出 containerd 独立于 docker，可以通过 binary 和各发行版的软件包管理部署

# containerd 流程
- 单纯的 containerd
![](/static/images/k8s/containerd/containerd-run-container-1.png)

    > [ctr](https://github.com/containerd/containerd/tree/main/cmd/ctr) 是 containerd 的命令行工具，属于 containerd 代码中的一部分，可以通过 ctr 或者自己的程序来调用 containerd 创建容器

- kubernetes 中的 containerd
![](/static/images/k8s/containerd/containerd-run-container-2.png)
    > containerd 实现了 k8s 的 [CRI](https://github.com/containerd/containerd/tree/main/pkg/cri) 接口，提供容器运行时核心功能，如镜像管理、容器管理等，也就是说 containerd 同样是一个 k8s CRI 的实现，可以使用 k8s提供的 [cri-tools](https://github.com/kubernetes-sigs/cri-tools) 中的 crictl(CLI for kubelet CRI) 命令行工具与 containerd 的 CRI 实现交互

# 安装 containerd
## 与 containerd 交互的工具
Name      | Community             | API    | Target             | Web site                                    |
----------|-----------------------|------- | -------------------|---------------------------------------------|
`ctr`     | containerd            | Native | For debugging only | (None, see `ctr --help` to learn the usage) |
`nerdctl` | containerd (non-core) | Native | General-purpose    | https://github.com/containerd/nerdctl       |
`crictl`  | Kubernetes SIG-node   | CRI    | For debugging only | https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md |


## binaries 方式

从 [containerd 流程](#containerd-流程)中可以分析出所需要部署的软件，[containerd](https://github.com/containerd/containerd/releases), [runc](https://github.com/opencontainers/runc/releases),这里使用 ubuntu 20.04 amd64 为例


软件|版本|架构
---|---|---
containerd|[v1.6.8](https://github.com/containerd/containerd/releases/download/v1.6.8/containerd-1.6.8-linux-amd64.tar.gz)|amd64
runc|[v1.1.4](https://github.com/opencontainers/runc/releases/download/v1.1.4/runc.amd64)|amd64

```shell
### 简单粗暴，所有的操作都是 root(仅限于测试环境，生产或需要的环境还是非 root) 用户，假设上述的软件包已经下载到本地了
# mkdir -p /usr/local/containerd
# tar -zxf  containerd-1.6.8-linux-amd64.tar.gz -C /usr/local/containerd/
# install -m 755 runc.amd64 /usr/local/containerd/bin/runc
# tree /usr/local/containerd/
/usr/local/containerd/
└── bin
    ├── containerd
    ├── containerd-shim
    ├── containerd-shim-runc-v1
    ├── containerd-shim-runc-v2
    ├── containerd-stress
    ├── ctr
    └── runc

1 directory, 7 files
# ln -s /usr/local/containerd/bin/* /usr/local/bin/
```

- [containerd.service](https://github.com/containerd/containerd/blob/main/containerd.service)
```shell
# cat <<EOF | tee /usr/lib/systemd/system/containerd.service
# Copyright The containerd Authors.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target local-fs.target

[Service]
#uncomment to enable the experimental sbservice (sandboxed) version of containerd/cri integration
#Environment="ENABLE_CRI_SANDBOXES=sandboxed"
ExecStartPre=-/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd

Type=notify
Delegate=yes
KillMode=process
Restart=always
RestartSec=5
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target

EOF
# systemctl daemon-reload
# systemctl enable --now containerd
# systemctl status containerd --lines 1
● containerd.service - containerd container runtime
     Loaded: loaded (/lib/systemd/system/containerd.service; enabled; vendor preset: enabled)
     Active: active (running) since Sun 2022-10-09 07:24:30 UTC; 21s ago
       Docs: https://containerd.io
    Process: 883269 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 883311 (containerd)
      Tasks: 11
     Memory: 20.8M
     CGroup: /system.slice/containerd.service
             └─883311 /usr/local/bin/containerd

Oct 09 07:24:30 sky systemd[1]: Started containerd container runtime.

```

## 软件包管理方式
参考 [docker](https://docs.docker.com/engine/install/) 中 containerd.io 的部署

## 测试

- 参看版本
```shell
# ctr version
Client:
  Version:  v1.6.8
  Revision: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
  Go version: go1.17.13

Server:
  Version:  v1.6.8
  Revision: 9cd3357b7fd7218e4aec3eae239db1f68a5a6ec6
  UUID: 8e8c7690-e13d-4ab8-a3ed-44dfab717631

```
- pull 镜像
```shell
# ctr images pull docker.io/library/alpine:3.14
docker.io/library/alpine:3.14:                                                    resolved       |++++++++++++++++++++++++++++++++++++++| 
index-sha256:1ab24b3b99320975cca71716a7475a65d263d0b6b604d9d14ce08f7a3f67595c:    done           |++++++++++++++++++++++++++++++++++++++| 
manifest-sha256:92d13cc58a46e012300ef49924edc56de5642ada25c9a457dce4a6db59892650: done           |++++++++++++++++++++++++++++++++++++++| 
config-sha256:dd53f409bf0bd55eac632f9e694fd190244fef5854a428bf3ae1e2b636577623:   done           |++++++++++++++++++++++++++++++++++++++| 
layer-sha256:c7ed990a2339ee598662849de4f56e2241399f5a32340c8c4a7bbd5378a12b5f:    done           |++++++++++++++++++++++++++++++++++++++| 
elapsed: 8.0 s                                                                    total:  2.7 Mi (345.5 KiB/s)                                     
unpacking linux/amd64 sha256:1ab24b3b99320975cca71716a7475a65d263d0b6b604d9d14ce08f7a3f67595c...
done: 149.29893ms
```
- 查看镜像
```shell
# ctr images list
REF                           TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
docker.io/library/alpine:3.14 application/vnd.docker.distribution.manifest.list.v2+json sha256:1ab24b3b99320975cca71716a7475a65d263d0b6b604d9d14ce08f7a3f67595c 2.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -    
```
- 运行容器
```shell
# ctr run -d docker.io/library/alpine:3.14 alpine
```
- 查看容器(container和task):
> 在 containerd 中，container 和 task 是分离的，container 描述的是容器分配和附加资源的元数据对象，是静态内容，task 是系统上一个活动的、正在运行的进程。 task 应该在每次运行后删除，而 container 可以被多次使用、更新和查询。这点和 docker 中 container 定义是不一样的。
```shell
# ctr container ls
CONTAINER    IMAGE                            RUNTIME                  
alpine       docker.io/library/alpine:3.14    io.containerd.runc.v2   
# ctr task ls
TASK      PID      STATUS    
alpine    86553    RUNNING
```
- 其他
> 可以看出 containerd 中是存在 namespace 概念的，这样可以将不同业务和应用进行隔离
```shell
### 查看系统进程
# ps -elf | grep alpine
0 S root       86533       1  0  80   0 - 178164 futex_ 15:40 ?       00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace default -id alpine -address /run/containerd/containerd.sock
### 查看存在的 namespaces
# ctr namespaces ls
NAME    LABELS 
default        
moby           
### 查看 namespace 中的镜像
# ctr -n default images ls
REF                           TYPE                                                      DIGEST                                                                  SIZE    PLATFORMS                                                                                LABELS 
docker.io/library/alpine:3.14 application/vnd.docker.distribution.manifest.list.v2+json sha256:1ab24b3b99320975cca71716a7475a65d263d0b6b604d9d14ce08f7a3f67595c 2.7 MiB linux/386,linux/amd64,linux/arm/v6,linux/arm/v7,linux/arm64/v8,linux/ppc64le,linux/s390x -      
# ctr -n moby images ls
REF TYPE DIGEST SIZE PLATFORMS LABELS

```

# Thanks
[Getting started with containerd](hhttps://github.com/containerd/containerd/blob/main/docs/getting-started.md)  
[部署容器运行时 Containerd](https://blog.frognew.com/2021/04/relearning-container-02.html)