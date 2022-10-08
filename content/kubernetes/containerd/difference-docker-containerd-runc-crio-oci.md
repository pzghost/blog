---
title: "Docker, Kubernetes, CRI, OCI, Containerd, Runc 之间的渊源"
isCJKLanguage: true
date: 2022-10-08T15:30:16+08:00
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
Docker拉开了容器的序幕，但不久之后，容器领域又出现了许多新的工具、标准和首字母缩写词。那么，"docker "到底是什么？"CRI "和 "OCI "这样的术语又是什么意思？


# 术语
- OCI(Open Container Initiative): 发布 containers 以及 imaegs 的规范
- CRI(Container Runtime Interface): 定义 Kubernetes 和 container runtime 之间的 API

# 生态圈
![](/static/images/k8s/containerd/container-ecosystem.drawio.webp)

# Docker

- docker CLI - 一个命令行界面程序。用户通过 docker CLI 命令来创建和管理 Docker 容器
- containerd - 一个管理和运行容器的守护程序。它拉取和存储请求的镜像、控制容器的生命周期
- runC - 实际创建和运行容器的 OCI 兼容的实现

![](/static/images/k8s/containerd/container-ecosystem-docker.drawio.webp)

# Docker images
Docker 镜像是包含了应用程序以及其所需的库、工具和其他依赖的一个只读的模板。当用户在 Docker 中发出运行命令时，镜像模板被用来部署一个应用容器。Docker 镜像是通过 Dockerfile 创建的，Dockerfile 是一个包含镜像信息的文本文件，docker build 使用 Dockerfile 和一个上下文来创建镜像。(注：Docker 镜像，实际上是以 OCI 标准格式打包的镜像。)

# Dockershim
> Docker 和 Kubernetes 之间是什么样的关系呢？

CRI 是允许 Kubernetes 与其他 Container runtime 通信的插件。但是 Docker 并没有实现 CRI， 所以 Kubernetes 包括一个叫做 dockershim 的组件， 用来桥接 kubernetes 和 docker 之间的通信

![](/static/images/k8s/containerd/container-ecosystem-dockershim.png)

# CRI
> Container Runtime Interface

- CRI 是 Kubernetes 用来控制创建和管理不同的 container runtime 的协议
- CRI 是对任何种类的 container runtime 的抽象。因此，CRI 使 Kubernetes更容易使用不同的 container runtime
- CRI 使 Kubernetes 不需要添加对每个 container runtime 的支持，CRI API 描述了 Kubernetes 如何与每个 container runtime 的进行交互。因此，container runtime 只要遵守 CRI API

![](/static/images/k8s/containerd/container-ecosystem-cri.drawio.webp)

# Containerd
containerd 是一个从 Docker 项目中分离出来的并实现了 CRI 规范的守护进程。Kubernetes 可以直接通过 containerd 的 cri 插件和 containerd 通信

![](/static/images/k8s/containerd/container-ecosystem-cri.png)

# CRI-O
CRI-O 是由 Redat,IBM,Intel,SUSE 等公司联合开发出的另一款 container runtime.

# OCI
> Open Container Initiative

OCI 是由一些技术公司在 2015 年建立了的，OCI 的目的是为容器格式和运行时创建标准。目前，OCI有两个规范。

- image-spec --镜像规格，概述了如何创建一个符合 OCI 标准的镜像
- runtime-spec--解开文件系统包的运行时规范

# RunC
> runC 是 OCI 的规格实现，为容器提供了所有的低级别功能，与 Linux 的 namespaces 和 cgroups 等进行交互，使用这些特性来创建和运行容器进程。

# Thanks 
[The differences between Docker, containerd, CRI-O and runc](https://www.tutorialworks.com/difference-docker-containerd-runc-crio-oci/)  
[Docker vs containerd vs CRI-O](https://phoenixnap.com/kb/docker-vs-containerd-vs-cri-o)  
[containerd](https://github.com/containerd/containerd)