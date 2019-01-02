---
title: containerd原生支持Kubernetes
update: 2018-03-21 20:25:47
tags:
- container
- kubernetes
categories:
- cloud
---



苗艳强，闫学安

ZTE 工程师，主要从事容器相关开源工作，Kubernetes 社区 member, containerd 社区 member

课程详情：

**演讲主题：**

目前跟 Kubernetes 对接的默认容器运行时是 docker，然而 docker 从 1.12 版本开始加入了 swarm 功能，随着版本的演进，功能越来越庞大，已经成为一个跟 Kubernetes 同级甚至高一级的编排工具，且前不久又宣布在编排测要无缝对接 Kubernetes，因此，docker 作为 Kubernetes 默认容器运行时的位子必将被其他工具代替。本次分享首先简单介绍 Kubernetes 的运行时接口规范，然后着重为大家分享一款新的容器运行时工具 containerd，包括项目介绍，功能组成等，以及使其原生对接 Kubernetes 的插件 cri-containerd。

**纲要/提纲：**

1. Kubernetes 容器容器运行时接口（CRI）

2. containerd 项目介绍

3. containerd 的 CRI 实现

-------------------------
Containerd的CRI实现
kubelet -> cri-containerd -> containerd