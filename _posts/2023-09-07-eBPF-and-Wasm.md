---
title: "eBPF and Wasm: Exploring the Future of the Service Mesh Data Plane"
categories:
  - blog
tags:
  - eBPF
  - Wasm
---

# eBPF and Wasm: 探索服务网格数据平面的未来

-----------

### 译者序

本文来自2022年[Vivian Hu](https://www.infoq.com/profile/Vivian-Hu/)的这篇文章：eBPF and Wasm: Exploring the Future of the Service Mesh Data Plane。主要介绍了eBPF和Wasm技术，探索服务网格的未来。

**译者水平有限，不免存在遗漏或错误之处。如有疑问，敬请查阅原文。**

以下是译文。

## 1 Sidecar 的概念

**什么是sidecar服务网格？**

服务网格（DP）负责网格内的服务通信，并可以通过单独的专用基础架构层提供负载均衡、加密和故障恢复。sidecar proxy附加到网格控制面（CP），所有的流量通过sidecar过滤，该代理作为自己的基础架构层运行。

![whatis-sidecar_proxy](https://github.com/Charonxin/charonxin.github.io/assets/91298205/8767f830-186a-4016-96a5-561099f198cb)

大概sidecar代理就是过滤包的这样一个东西。

如果使用kubernetes监视应用程序，则可以将容器组合在一个共享命名空间的Pod中，然后一个单独的sidecar容器可以可视化同一Pod中每个容器的运行方式。

**优点：**Sidecar使开发人员将微服务或容器分离，快速有效地监控和维护其应用程序。包括降低代码复杂性、减少重复代码、降低程序之间的耦合。

## 2 eBPF重塑sidecar时代 —— sidecarless

2021年12月，Cilium项目宣布了Cilium Service Mesh的beta测试计划。基于谷歌云、eBPF的Kubernetes服务（GKS）数据平面开创的概念，一年前于2020年8月宣布。它旨在推广“消灭sidecar”的概念。扩展了eBPF产品，以处理服务网格的许多sidecar proxy功能。

**一个服务网格，必须具有以下功能：**

- 弹性连接：服务到服务通信必须能够跨越边界，例如云、集群和本地。通信必须具有复原能力和容错能力。 
- L7 流量管理：负载平衡、速率限制和弹性必须是 L7 感知的（HTTP、REST、gRPC、WebSocket 等）。
- 基于身份的安全性：依靠网络标识符来实现安全性已不再足够，发送和接收服务都必须能够基于身份而不是网络标识符相互进行身份验证。 
- 可观测性和跟踪：跟踪和指标形式的可观测性对于理解、监视和排除应用程序稳定性、性能和可用性问题至关重要。 
- 透明性：该功能必须以透明的方式提供给应用程序，即不需要更改应用程序代码。

应用程序习惯于发布自己的TCP / IP堆栈！**服务网格正在演变为内核责任**，就像网络堆栈一样。

![02](https://github.com/Charonxin/charonxin.github.io/assets/91298205/21022d2e-4aaf-4316-b575-b4a748c3edec)

但后来演变到了sidecar模式的模型

![微信截图_20230907223920](https://github.com/Charonxin/charonxin.github.io/assets/91298205/9336067b-8fef-475b-ae57-a8f4f6e366cd)

此模型的缺点是大量的代理、许多额外的网络连接以及将网络流量馈送到代理中的复杂重定向逻辑。

几十年来，在应用程序之间提供安全可靠的连接一直是操作系统的责任。早期Unix和Linux时代的**TCP Wrappers**和**tcpd**。TCPD 允许用户透明地向应用程序添加日志记录、访问控制、主机名验证和欺骗保护，而无需对其进行修改。它使用了**libwrap**，并且，应用程序以前链接了这个库来提供这些功能。tcpd 带来的是能够透明地将功能添加到现有应用程序而无需修改它们。最终，**所有这些功能都进入了Linux，并以更高效，更强大的方式提供给所有应用程序**。今天，这已经演变成我们所知道的**iptables**。

![微信截图_20230907223920](https://github.com/Charonxin/charonxin.github.io/assets/91298205/e0bf6c0f-2375-4087-aee7-f6ed36dc7675)

然而，iptables 显然不适合解决现代应用程序的连接性、安全性和可观测性的要求，因为它只在网络级别运行，缺乏对应用层的了解。现在，我们已经到了在操作系统中原生支持此功能以实现最佳透明度、效率和安全性的地步。

## 3 扩展内核命名空间的概念

Linux 内核已经具有**共享通用功能**的概念，并使其可用于许多应用程序。这个概念被称为**命名空间**，它构成了我们今天所知的容器技术的基础。命名空间（内核类型，而不是 Kubernetes 版本）存在于各种抽象中，包括**文件系统、用户管理、挂载的设备、进程、网络**等等。这允许单个容器具有不同的文件系统视图、不同的用户集，并**允许多个容器绑定到单个主机上的同一网络端口**。这个概念在 cgroups 的帮助下进行了扩展，以管理 CPU、内存和网络等资源应用资源。从云原生应用程序开发人员的角度来看，cgroups 和资源紧密集成到我们称为“容器”的概念中。

如果我们将服务网格视为操作系统的责任，那么它必须符合并集成命名空间和 cgroups 的概念，像这样：

![pic-4](https://github.com/Charonxin/charonxin.github.io/assets/91298205/8b964105-4bff-49d2-9862-4b09cbf796d1)

## 4 使用 eBPF 解锁内核服务网格

**为什么我们之前没有在内核中创建服务网格？**

有些人半开玩笑地说 **kube-proxy** 是原始的服务网格。这有一定的道理。Kube-proxy 说明 Linux 内核在**依赖使用 iptables 实现**的传统基于网络的功能的同时，可以多么接近实现服务网格。但是，这还不够，缺少L7上下文。**Kube-proxy 只在网络数据包级别运行**。现代应用程序需要 L7 流量管理、跟踪、身份验证和其他可靠性保证。**Kube-proxy 无法在网络级别提供此功能**。

**eBPF改变了这个现状**。它允许动态扩展Linux内核的功能。我们一直在使用 eBPF for Cilium 来构建一个高效的网络、安全性和可观测性数据路径，该数据路径将自身直接嵌入到 Linux 内核中。

eBPF具有巨大的优势。eBPF 代码可以在**运行时插入到现有的 Linux 内核**中，类似于 Linux 内核模块，但与内核模块不同的是，它可以以**安全和可移植**的方式完成。这允许 eBPF 实施随着服务网格社区的不断发展而继续发展。新的内核版本需要数年时间才能到达用户手中。eBPF 是使 Linux 内核能够跟上**快速发展的云原生技术堆栈**的关键技术。

## 5 eBPF and Wasm 

eBPF的许多问题都与它是一个内核技术有关，因此必须有**安全限制**。有没有办法使用空间技术将应用程序的代理逻辑**合并到数据平面**中，而**不会降低性能**？事实证明，WebAssembly（Wasm）可能就是这样的选择。Wasm 运行时可以**安全地隔离和执行用户空间代码**，性能接近本机。

2021 年 12 月，WasmEdge 贡献者社区证明，基于 WasmEdge 的微服务可以与 Dapr 和 Linkerd sidecar 配合使用，作为 Linux 容器的替代方案。与 Linux 容器应用程序相比，WebAssembly 微服务消耗 1% 的资源，冷启动的时间只有 1%。

## 总结

eBPF 和 Wasm 是服务网格应用程序的新成员，可在数据平面中实现高性能。它们仍然是新兴技术，但有可能成为微服务生态系统中当今Linux容器的替代品或补充。

**如有不当恳请指正。**
